# PostgreSQL proxies: PgBouncer (L4) vs ProxySQL (L7)

Hands-on lab for the blog post *"Layer 4 vs. Layer 7 Database Proxies: Scaling PostgreSQL
Under Mass Concurrency"* (Hossein Askari, Substack).

| File | What it deploys |
|---|---|
| `01-postgres.yaml` | Namespace `pg-proxy`, Postgres 17 primary + streaming replica, `max_connections=50` |
| `02-pgbouncer.yaml` | PgBouncer 1.26 (2 pods), transaction pooling |
| `03-proxysql.yaml` | ProxySQL 3.0 with native PostgreSQL: R/W split + query cache |
| `04-hybrid-pgbouncer-proxysql.yaml` | PgBouncer at the edge → ProxySQL → Postgres |

```
                 ┌──────────────── pgbouncer :6432 ───────────────┐
                 │  appdb    → pg-primary                         │  (app picks R/W)
 clients ────────┤  appdb_ro → pg-replica                         │
                 │                                                │
                 ├──────────────── proxysql :6133 ────────────────┤
                 │  SELECT            → HG20 pg-replica           │  (proxy picks R/W)
                 │  SELECT FOR UPDATE → HG10 pg-primary           │
                 │  everything else   → HG10 pg-primary           │
                 │                                                │
                 └── pgbouncer-edge :6432 → proxysql :6133 ───────┘  (hybrid)
```

Lab credentials (in Secrets, do not reuse): app `app/app-pass`, ProxySQL admin `radmin/radmin-pass`.

---

## 1. Deploy

```bash
kubectl apply -f 01-postgres.yaml
kubectl -n pg-proxy rollout status sts/pg-primary
kubectl -n pg-proxy rollout status sts/pg-replica

kubectl apply -f 02-pgbouncer.yaml -f 03-proxysql.yaml -f 04-hybrid-pgbouncer-proxysql.yaml
kubectl -n pg-proxy get pods
```

Check replication:

```bash
kubectl -n pg-proxy exec pg-primary-0 -- psql -U postgres -c "select client_addr, state from pg_stat_replication"
kubectl -n pg-proxy exec pg-replica-0 -c postgres -- psql -U postgres -c "select pg_is_in_recovery()"   # t
```

Helpers used below (the client runs inside the primary pod, which already has `psql`):

```bash
q() {  # q <host> <port> <db> <sql>
  kubectl -n pg-proxy exec pg-primary-0 -- env PGPASSWORD=app-pass \
    psql -h "$1" -p "$2" -U app -d "$3" -At -c "$4"
}
admin() {  # ProxySQL admin, speaks the Postgres protocol on 6132
  kubectl -n pg-proxy exec pg-primary-0 -- env PGPASSWORD=radmin-pass \
    psql -h proxysql -p 6132 -U radmin -d admin -c "$1"
}
```

## 2. PgBouncer: L4, no SQL awareness

```bash
q pgbouncer 6432 appdb    "select inet_server_addr(), pg_is_in_recovery()"   # primary, f
q pgbouncer 6432 appdb_ro "select inet_server_addr(), pg_is_in_recovery()"   # replica, t
```

PgBouncer never looks at the query. The **database name** picks the backend, so
read/write splitting is the application's job.

Pool internals via the admin console:

```bash
q pgbouncer 6432 pgbouncer "SHOW POOLS"    # cl_active / cl_waiting / sv_active / sv_idle
q pgbouncer 6432 pgbouncer "SHOW STATS"
```

## 3. ProxySQL: L7, transparent R/W split

```bash
q proxysql 6133 appdb "CREATE TABLE IF NOT EXISTS orders(id serial primary key, item text)"
q proxysql 6133 appdb "INSERT INTO orders(item) VALUES ('book')"
q proxysql 6133 appdb "SELECT pg_is_in_recovery()"              # t  -> replica
q proxysql 6133 appdb "SELECT pg_is_in_recovery() FOR UPDATE"   # f  -> primary
q proxysql 6133 appdb "BEGIN; INSERT INTO orders(item) VALUES ('x'); SELECT pg_is_in_recovery(); COMMIT;"
#   f -> a transaction stays on the primary, even its SELECTs
```

Where did each query go?

```bash
admin "SELECT hostgroup, digest_text, count_star FROM stats_pgsql_query_digest ORDER BY count_star DESC"
admin "SELECT * FROM runtime_pgsql_servers"
admin "SELECT hostname, ping_error FROM monitor.pgsql_server_ping_log ORDER BY time_start_us DESC LIMIT 4"
```

### Query cache (and its trade-off)

Rule 5 caches any query containing `/* cache */` for 5s:

```bash
q proxysql 6133 appdb "SELECT count(*) /* cache */ FROM orders"   # N  (miss, stored)
q proxysql 6133 appdb "INSERT INTO orders(item) VALUES ('pen')"
q proxysql 6133 appdb "SELECT count(*) /* cache */ FROM orders"   # N  <- STALE for up to 5s
q proxysql 6133 appdb "SELECT count(*) FROM orders"               # N+1 (uncached)
admin "SELECT * FROM stats_pgsql_global WHERE variable_name LIKE 'Query_Cache_count%'"
```

> The cache rule uses `match_pattern` (raw SQL), not `match_digest`: digests strip comments.

### Change rules live, with no restart

```bash
# Emergency: block a destructive query pattern
admin "INSERT INTO pgsql_query_rules (rule_id, active, match_digest, error_msg, apply)
       VALUES (2, 1, '^DELETE FROM orders$', 'blocked by DBA', 1)"
admin "LOAD PGSQL QUERY RULES TO RUNTIME"
q proxysql 6133 appdb "DELETE FROM orders"     # ERROR: blocked by DBA
```

Runtime changes live only in memory. The datadir is an `emptyDir`, so a pod restart
reloads `proxysql.cnf` from the Secret. Put permanent changes in `03-proxysql.yaml`.

## 4. Hybrid: PgBouncer edge → ProxySQL

```bash
q pgbouncer-edge 6432 appdb "SELECT pg_is_in_recovery()"   # t, routed by ProxySQL
q pgbouncer-edge 6432 appdb "INSERT INTO orders(item) VALUES ('hybrid') RETURNING id"
```

## 5. Connection storm with pgbench

```bash
bench() {  # bench <host> <port> [pgbench args...]
  local h=$1 p=$2; shift 2
  kubectl -n pg-proxy run "pgbench-$h-$RANDOM" --rm -i --restart=Never \
    --image=postgres:17.11 --env=PGPASSWORD=app-pass -- \
    pgbench -h "$h" -p "$p" -U app "$@" appdb
}
backends() {  # run in a 2nd terminal while bench is running
  for n in primary replica; do
    kubectl -n pg-proxy exec pg-$n-0 -c postgres -- psql -U postgres -At \
      -c "select '$n: '||count(*) from pg_stat_activity where usename='app'"
  done
}

bench pg-primary 5432 -i -s 10                      # create pgbench tables (once)

bench pg-primary     5432 -c 200 -j 4 -T 15 -S      # direct: FAILS, too many clients
bench pgbouncer      6432 -c 200 -j 4 -T 15 -S      # L4 pool
bench proxysql       6133 -c 200 -j 4 -T 15 -S      # L7, reads go to the replica
bench pgbouncer-edge 6432 -c 200 -j 4 -T 15 -S      # hybrid
```

Results on a 3-node k3d cluster (laptop), 200 clients, select-only:

| Path | Result | tps | avg latency | app backends (primary / replica) |
|---|---|---|---|---|
| direct | `remaining connection slots are reserved` | – | – | 50 max |
| pgbouncer | 0 failed | ~129k | 1.5 ms | pooled on primary / 0 |
| proxysql | 0 failed | ~85k | 2.3 ms | – / capped at HG20 `max_connections` |
| pgbouncer-edge → proxysql | 0 failed | ~47k | 4.3 ms | – / capped at HG20 `max_connections` |

This matches the blog: L4 has the lowest overhead, L7 adds parse cost but moves
reads off the primary, and every hop adds latency. With PgBouncer and ProxySQL each
hit by 300 read/write clients **at the same time** (600 total), the primary held
about 35 backends and had zero failed transactions.

### Connection budget (primary `max_connections=50`)

```
pgbouncer   2 pods x max_db_connections 12 = 24
proxysql    HG10 max_connections           = 15
                                       total = 39   (+3 reserved for superuser)
```

Pooler limits apply **per pod**. Scaling the PgBouncer Deployment multiplies its
share, so re-check this budget whenever you change replicas.

## 6. Things to try

- `pool_mode = session` in `02-pgbouncer.yaml`, then rerun bench. Clients now hold server
  connections, so `cl_waiting` climbs in `SHOW POOLS`.
- Use `pgsql_replication_hostgroups` so ProxySQL detects the primary from
  `pg_is_in_recovery()` instead of using static hostgroups.
- Scale the replica (`kubectl -n pg-proxy scale sts/pg-replica --replicas=2`). ProxySQL can
  load-balance per pod, but only if each pod has its own DNS name. Make the `pg-replica`
  Service headless (`clusterIP: None`) and list `pg-replica-0.pg-replica.pg-proxy.svc.cluster.local`
  and `pg-replica-1...` separately in `pgsql_servers` HG20.

## Cleanup

```bash
kubectl delete ns pg-proxy
```
