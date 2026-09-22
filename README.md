# dokku cassandra

Cassandra plugin for dokku. Currently defaults to installing [cassandra 5.0.9](https://hub.docker.com/_/cassandra/).

Not an official `dokku` org plugin — see [docs/README.md](docs/README.md) and the "About this plugin" section below for how it differs from redis/mongo/postgres/mysql, which are.

## Requirements

- dokku 0.19.x+
- docker 1.8.x

## Installation

```shell
sudo dokku plugin:install https://github.com/<your-user>/dokku-cassandra.git --name cassandra
```

## Commands

```
cassandra:admin-console <service>                    # open a cqlsh shell against the service (alias of connect)
cassandra:app-links <app>                             # list all cassandra service links for a given app
cassandra:clone <service> <new-service> [--clone-flags...] # create container <new-service> then copy data from <service> into <new-service>
cassandra:connect <service>                            # connect to the service via cqlsh
cassandra:create <service> [--create-flags...]         # create a cassandra service
cassandra:destroy <service> [-f|--force]                # delete the cassandra service/data/container if there are no links left
cassandra:enter <service> [cmd...]                      # enter or run a command in a running cassandra service container
cassandra:exists <service>                              # check if the cassandra service exists
cassandra:export <service>                              # export a snapshot of the cassandra service data directory
cassandra:expose <service> <ports...>                   # expose a cassandra service on custom host:port
cassandra:import <service>                              # import a snapshot into the cassandra service data directory
cassandra:info <service> [--flag]                       # print the service information
cassandra:link <service> <app> [--link-flags...]        # link the cassandra service to the app
cassandra:linked <service> <app>                        # check if the cassandra service is linked to an app
cassandra:links <service>                                # list all apps linked to the cassandra service
cassandra:list                                           # list all cassandra services
cassandra:logs <service> [-t|--tail] [n]                 # print the most recent log(s) for this service
cassandra:pause <service>                                # pause a running cassandra service
cassandra:promote <service> <app>                        # promote service <service> as CASSANDRA_URL in <app>
cassandra:restart <service>                              # graceful shutdown and restart of the cassandra service container
cassandra:set <service> <key> <value>                    # set or clear a property for a service
cassandra:start <service>                                # start a previously stopped cassandra service
cassandra:stop <service>                                 # stop a running cassandra service and remove the container
cassandra:unexpose <service>                             # unexpose a previously exposed cassandra service
cassandra:unlink <service> <app>                         # unlink the cassandra service from the app
cassandra:upgrade <service> [--upgrade-flags...]         # upgrade service <service> to the specified image/version
```

## Usage

Help for any command can be displayed by passing it to `cassandra:help`, e.g. `dokku cassandra:help create`.

### create a cassandra service

```shell
# usage
dokku cassandra:create <service> [--create-flags...]
```

flags:

- `-i|--image <string>`: the image name to start the service with (default: `cassandra`)
- `-I|--image-version <string>`: the image version to start the service with (default: `5.0.9`)
- `-m|--memory <int>`: container memory limit in megabytes (default: unlimited)
- `-c|--config-options <string>`: extra JVM flags, passed through as `JVM_EXTRA_OPTS`
- `-C|--custom-env <string>`: semi-colon delimited environment variables to start the service with
- `-d|--cluster-name <string>`: the Cassandra cluster name (default: the service name)
- `-N|--initial-network <string>`: the initial network to attach the service to
- `-P|--post-create-network <strings>`: a comma-separated list of networks to attach the service container to after service creation
- `-S|--post-start-network <strings>`: a comma-separated list of networks to attach the service container to after service start

Create a cassandra service named lollipop:

```shell
dokku cassandra:create lollipop
```

A fresh container can take one to three minutes to become reachable (JVM start plus bootstrap); the command waits for `cqlsh` to answer before returning.

### print the service information

```shell
# usage
dokku cassandra:info <service> [--single-info-flag]
```

flags: `--config-options`, `--data-dir`, `--dsn`, `--exposed-ports`, `--id`, `--internal-ip`, `--initial-network`, `--links`, `--post-create-network`, `--post-start-network`, `--service-root`, `--status`, `--version`

```shell
dokku cassandra:info lollipop
dokku cassandra:info lollipop --dsn
```

### link the service to an app

```shell
# usage
dokku cassandra:link <service> <app> [--link-flags...]
```

flags:

- `-a|--alias <string>`: an alternative alias to use for linking to an app via environment variable
- `-q|--querystring <string>`: ampersand delimited querystring arguments to append to the service link
- `-n|--no-restart`: do not restart the app on link (default: restarts)

```shell
dokku cassandra:link lollipop playground
```

This sets `CASSANDRA_URL` on the app to `cassandra://dokku-cassandra-lollipop:9042/lollipop` — see [docs/README.md](docs/README.md) for why there are no credentials in it and why the keyspace at the end is not created for you.

### expose a service on a host port

```shell
# usage
dokku cassandra:expose <service> <ports...>
```

```shell
dokku cassandra:expose lollipop 9042
dokku cassandra:expose lollipop 127.0.0.1:9042
```

### export / import

```shell
dokku cassandra:export lollipop > backup.tar
dokku cassandra:import lollipop < backup.tar
```

`export` flushes memtables to disk and streams the whole data directory as a tar; `import` stops the container, replaces the data directory, and starts it back up. Both move the raw sstables, not a per-keyspace logical dump — see [docs/README.md](docs/README.md).

### clone a service

```shell
dokku cassandra:clone lollipop lollipop-2
```

Creates `lollipop-2` and copies `lollipop`'s data into it (implemented as an export piped into an import).

### upgrade a service

```shell
# usage
dokku cassandra:upgrade <service> [--upgrade-flags...]
```

Takes the same `-i|--image`, `-I|--image-version`, `-m|--memory`, `-c|--config-options`, `-C|--custom-env` flags as `create`. Recreates the container against the new image/settings; data is untouched since it lives on the bind-mounted volume.

```shell
dokku cassandra:upgrade lollipop --image-version 5.0.9
```

## About this plugin

Dokku's own datastore plugins (redis, mongo, postgres, mysql, mariadb, elasticsearch, and a few more) are thin wrappers around a shared, closed-source binary called [`dokku-datastore`](https://github.com/dokku/dokku-datastore) that ships every one of those engines' definitions built in. That binary was checked directly (`dokku-datastore generate cassandra`) while building this plugin, and it rejects any type name it does not already know — Cassandra isn't one of the ~20 it embeds, and there is no supported way for a third-party plugin to register a new one.

So this plugin implements its service lifecycle itself, in bash, the way every community datastore plugin does (and the way `dokku-redis`/`dokku-postgres`/etc. themselves did before their move to `dokku-datastore`). The file layout mirrors the current `dokku-mongo`/`dokku-redis` repos as closely as that constraint allows: `plugin.toml`, `config`, `commands`, one file per subcommand under `subcommands/`, the same root-level app-lifecycle triggers (`pre-start`, `pre-restore`, `pre-delete`, `post-app-clone-setup`, `post-app-rename-setup`, `service-list`), and a `docs/README.md` for supplemental notes. What differs is `functions`/`common-functions`, which hold this plugin's own implementation instead of a call into `dokku-datastore`, and the lack of an `install` script, since there is no external binary for this plugin to fetch.

## Known limitations (v0.1)

See [docs/README.md](docs/README.md) for the full notes. In short: no authentication, no auto-created keyspace, single node per service, and `export`/`import` move raw sstables rather than a logical per-keyspace dump. S3 backup commands (`backup`, `backup-auth`, `backup-schedule`, ...), which the official plugins offer, are not implemented in this version.
