# Extra Documentation

The documentation in this folder is supplemental to using this plugin.

## Notes

- No authentication is configured. The official `cassandra` image has no environment variable that turns on `PasswordAuthenticator` the way `requirepass`/`MYSQL_PASSWORD`/`POSTGRES_PASSWORD` work for other datastores, and shipping a full custom `cassandra.yaml` just to flip that one key was judged not worth the added maintenance surface for a first version. Isolation is the docker network: a service is only reachable by containers linked to it, unless you run `cassandra:expose`.
- The keyspace named in the connection URL is **not** created for you. Unlike postgres/mysql/mongo, the base image has no env var for this either. Create it yourself over `cassandra:connect` before pointing an app at it, e.g. `CREATE KEYSPACE myapp WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};`.
- Each service is a single Cassandra node. `cassandra:create` does not form a multi-node cluster; it is meant for development, staging, or workloads that fit comfortably on one node.
- `cassandra:export`/`cassandra:import` flush and stream the whole data directory (data, commitlog, saved_caches, hints) rather than a single keyspace, since the image ships no per-keyspace dump tool comparable to `pg_dump`/`mysqldump`. Restoring cleanly requires the target service to run the same Cassandra version as the export.
- A fresh container can take one to three minutes to become reachable (JVM start plus bootstrap); `cassandra:create` and `cassandra:start` poll until `cqlsh` answers, up to `PLUGIN_STARTUP_TIMEOUT` attempts (default 180, at 2s apart).
