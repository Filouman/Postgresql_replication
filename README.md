## Guide to Set up PostgreSQL synchronous replication

### Requirements
- Two servers: primary and standby

- PostgreSQL installed (same version) on both

**Important:** Verify data directories, IP addresses, ports and adjust as needed.

<br>


### SETUP

### 1. Set Up the Primary PostgreSQL Server

Configure PostgreSQL for replication on the primary server.

Edit the PostgreSQL configuration file (`postgresql.conf`), usually located in `/etc/postgresql/xx/main/postgresql.conf` or a similar path.
```
sudo nano /etc/postgresql/17/main/postgresql.conf --linenumbers
```

Enable WAL Archiving: Set the following in `postgresql.conf` to enable listening and ensure all transactions are logged for replication:
```
listen_addresses = '*'
wal_level = replica
synchronous_commit = remote_apply
max_wal_senders = 5
wal_keep_size = 2000
synchronous_standby_names = 'standby1'      # name must match application_name on standby server
```


Allow Replication Connections: In the `pg_hba.conf` file, allow the standby server to connect:
```
sudo nano /etc/postgresql/17/main/pg_hba.conf --linenumbers
```
Add:
```
host    replication    replication_user    <standby_server_ip>/32    md5
```

Next, create a replication role:
```
psql -U postgres
CREATE ROLE replication_user REPLICATION LOGIN ENCRYPTED PASSWORD 'your_password';
```
Create a Physical Replication Slot (still in psql prompt):
```
SELECT * FROM pg_create_physical_replication_slot('standby_slot1');
```

Reload PostgreSQL
```
sudo systemctl restart postgresql
```

<br>

### 2. Set Up the Standby PostgreSQL Server


Stop PostgreSQL
```
sudo systemctl stop postgresql
```

Set the Connection Details in `postgresql.conf`
Edit `/etc/postgresql/17/main/postgresql.conf` (adjust path as needed) and add:
```
primary_conninfo = 'host=XXX.XXX.XXX.XX port=5432 user=replica_user password=your_password application_name=standby1'
primary_slot_name = 'standby_slot1'
```
*Adjust IP and port as needed to match connection details for the Primary server. Make sure the `application_name` matches the name in `synchronous_standby_names` on the Primary.


Backup Existing Data Directory, Create New Directory, and Set Permissions:
```
sudo mv /data/postgresdb/17/ /data/postgresdb/17.bak
sudo mkdir /data/postgresdb/17/
sudo chown postgres:postgres /data/postgresdb
sudo chmod 700 /data/postgresdb
```

Initialize Standby with Data from the Primary: On the Standby, use `pg_basebackup` to create an initial copy of the primary database:
```
pg_basebackup -h XXX.XXX.XXX.XX -D /data/postgresdb/17 -U replica_user -P -Xs -R
```
`-R` creates the `standby.signal` file and sets `primary_conninfo` automatically.
You’ll be prompted for the replication user’s password.

Manually Add `standby.signal` File
If `-R` wasn't used or you want to ensure it's there:
```
sudo touch /data/postgresdb/17/standby.signal
```
Start PostgreSQL on Standby server
```
sudo systemctl start postgresql
```

<br>

### 3. Monitoring and Testing
After configuring, restart both PostgreSQL servers. The Primary server will wait for acknowledgment from the Standby before confirming a transaction commit.

You can monitor the PostgreSQL main instance by the systemd journal and check the log output. You can exit the monitoring at any time with `Ctrl+C`
```
journalctl -fu postgresql
```

Monitor Logs:
```
sudo tail -f /var/log/postgresql/postgresql-17-main.log
```

On the Primary, check replication status:
```
SELECT pid, state, sync_state, application_name, client_addr
FROM pg_stat_replication;
```
