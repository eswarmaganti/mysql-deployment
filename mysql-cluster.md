# MySQL Cluster Architecture

## High-level Kubernetes Architecture
- `StatefulSet`
  - *Purpose*: Manages MYSQL Pods with stable network IDs and persistent storage
  - *Configuration*:
    - Replicas: 3 (Primary: 1 & Read Replicas: 2)
    - Ordered, graceful scaling & rolling updates -> pods are created / deleted in sequence.
    - provides sticky identity (`mysql-cluster-0`, `mysql-cluster-1`, & `mysql-cluster-2`).
- `PersistentVolumeClaim (PVC)`
  - *Purpose*: Ensures data survives Pod restarts, rescheduling, or node failures.
  - Each pod in StatefulSet gets its own PVC (data isolation)
  - Backed by a *StorageClass*
- `ConfigMap`
  - *Purpose*: Stores non-sensitive configs such as MySQL tuning, parameters, replication configs.
  - Example: `my.cnf` or replication settings
- `Secret`:
  - *Purpose*: Securely stores MySQL root password or replication user credentials.
  - Mounted as environment variables or files
- `Headless Service`:
  - *Purpose*: Provides stable DNS names for primary pod.
  - *Example*: `mysql-cluster-0`
  - Required for Pods to discover each other in the replication cluster.
- `ClusterIP Service`:
  - *Purpose*: Provides a single access point for apps inside the cluster.
  - instead of connecting to individual Pods, clients connect to `mysql-cluster-svc`.
- `Init Containers`:
  - *Purpose*: Run before the main MySQL container starts.
  - *Used for*:
    - Checking if a primary already exists.
    - Initializing the DB schema.
    - Setting up replication user & permissions
- `Sidecar Container` (xtrabackup):
  - *Purpose*: Handles backup * restore and replication setup.
  - Ensures new replicas are provisioned from a backup instead of a full sync.


## StatefulSet Deployment Architecture
- StatefulSet creates `mysql-cluster-0` first, waits Ready, then `mysql-cluster-1` then `mysql-cluster-2`
- For each pod, init container run (in order) before any app containers:
  - `init-mysql` -> runs first
  - `clone-mysql` -> runs second
  - Only if all init containers succeded will the main containers (`mysql-cluster + xtrabackup` sidecar) start.
- When the main containers start, readiness/liveness probe control when the pod is considered ready.

### `inti-mysql` init container:
- Runs `mysql:9` image and:
  - Parses pod ordinal from `$HOSTNAME` (e.g, `mysql-1` -> `1`)
  - Writes a unique MySQL `server-id` file into the shared `conf` (`/mnt/conf/server-id.cnf`) -- `server-id=100 + ordinal`
    - Why: each MySQL instance must have a unique `server-id` for replication.
  - Copies either `primary.cnf` (for ordinal 0) or `replica.cnf` (for others) from the ConfigMap into the same conf dir.
  - *Result*: each pod has its own config (primary vs replica) AND a unique server-id available at `/etc/mysql/conf.d` when container start.

### `clone-mysql` init container 
- *image*: `gcr.io/google-samples/xtrabackup:1.0`
- *Purpose*: **ensures** replicas start from a consistent snapshot of an existing peer.
- *Logic*:
  - if the data directory already contains MySQL (`/var/lib/mysql/mysql`) - skip (idempotent)
  - If this is the primary (ordinal == 0) -- skip
  - Otherwise:
    - connect to the previous pod (mysql-cluster-$(ordinal-1)-mysql-cluster) on port 3307 and receive an XtraBackup Stream:
    `ncat --recv-only mysql-cluster-$((ordianl-1)).mysql-cluster 3307 | xbsstream -x -C /var/lib/mysql`
    - Run `xtrabackup --prepare --target-dir=/var/lib/mysql` to make the backup consistent (apply logs)
- *Result*: A replica gets a prepared snapshot of data identical to the donor at a particular binlog position.

### `xtrabackup` sidecar container
- Start alongside the MySQL server. Exposes port 3307 and does two things.
  - A. `Prepare replication information`:
    - Looks for files lift by XtraBackup:
      - `xtrabackup_binlog_info` - contains: `<binlog-file> <postion> ` (if clone came directly from primary).
      - `xtrabackup_slave_info` - contains a `CHANGE MASTER TO` snippet (if clone came from a replica).
    - Builds `change_master_to.sql.in` (a partial `CHANGE MASTER TO` statement) using these files.
  - B. If `change_master_to.sql.in` exists:
    - Waits for MySQL to be accepting connections (`until mysql -h 127.0.0.1 -e "SELECT 1"`).
    - Run the below command to start replication from the correct position 
      ```commandline
      CHANGE MASTER TO ...
        MASTER_HOST='mysql-cluster-0.mysql-clister',
        MASTER_USER='root'
        MASTER_PASSWORD='',
        MASTER_CONNECT_RETRY=10;
      START SLAVE;
      ```
    - Renames the files so the same work avoids re-running on restarts.
  - C. Serve backups to future replicas
    - Starts a one-connection, keep-open `ncat` server:
      ```commandline
      ncat --listen --keep-open --send-only --max-conns=1 3307 -c "xtrabackup --backup --slave-info --stream=xbsstream --host=127.0.0.1 --user=root"
      ```
    - That command streams a hot backup (with slave/binlog metadata) to whomever connects (the next replica's `clone-mysql`)