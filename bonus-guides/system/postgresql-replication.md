# PostgreSQL Database Replication

 Write-Ahead Log (WAL) replication provides high availability and scalability by asynchronously copying data from a primary database server to one or more standby servers.  This ensures data redundancy and allows for failover in case of primary server failure.

{% hint style="warning" %}
Difficulty: Medium
{% endhint %}

## Streaming replication background

PostgreSQL has a feature called [streaming replication](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION) which provides the ability to continuously ship and apply the WAL records to some number of standby servers to keep them current.

The step that turns the file-based log-shipping standby into a streaming replication standby is setting the ```primary_conninfo``` setting to point to the primary server. ***This will be accomplished automatically when we perform the ```pg_basebackup``` command with the ```-R``` flag.***

### Syncronous replication

PostgreSQL streaming replication is asynchronous by default. If the primary server crashes then some transactions that were committed may not have been replicated to the standby server, causing data loss. The amount of data loss is proportional to the replication delay at the time of crashing.

Synchronous replication offers the ability to confirm that all changes made by a transaction have been transferred to one or more synchronous standby servers. This extends that standard level of durability offered by a transaction commit.

Once streaming replication has been configured, configuring synchronous replication requires only one additional configuration step: [synchronous_standby_names](https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-SYNCHRONOUS-STANDBY-NAMES) must be set to a non-empty value. ```synchronous_commit``` must also be set to ```on```, but since this is the default value, typically no change is required. This configuration will cause each commit to wait for confirmation that the standby has written the commit record to durable storage.

### Replication slots

Replication slots provide an automated way to ensure that the primary server does not remove WAL segments until they have been received by all standbys, and that the primary does not remove rows which could cause a recovery conflict even when the standby is disconnected. 

{hint style="info"}
**Caution**

Beware that replication slots can cause the server to retain so many WAL segments that they fill up the space allocated for ***pg_wal*** (in the event the standby server becomes unresponsive). [max_slot_wal_keep_size](https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-MAX-SLOT-WAL-KEEP-SIZE) can be used to limit the size of WAL files retained by replication slots.
{endhint}

## Preparation

### Stop all services using PostgreSQL

**IMPORTANT** - Stop LND and any additional services that rely on PostgreSQL (i.e. LNDg, AlbyHub)

```bash
sudo systemctl stop lnd
```

### Firewall

Configure primary server firewall to allow connections to PostgreSQL from the standby server

```bash
sudo ufw allow 5432/tcp comment 'allow PostgreSQL from anywhere'
```

<!---
### Set password for postgres user

You can set the password for the ```postgres``` user from the command line using the ```psql``` command-line utility. Since ```postgres``` is a special user, you need to connect as a user with sufficient privileges (like ```postgres`` itself, or another superuser) to change its password.

* Connect to psql as a privileged user:

```bash
sudo -u postgres psql
```

* Issue the ALTER ROLE command (replace ```'your_new_password'``` with the actual strong password you want to use.) :

```bash
* ALTER ROLE postgres WITH PASSWORD 'your_new_password';
```

* Verify the change - this will prompt you to enter the new password again, confirming it's set correctly:

```bash
\password postgres
```

* Exit psql:

```bash
\q
```

--->

## Primary server configuration

PostgreSQL configuration files are stored in the ```/etc/postgresql/<version>/main``` directory. For example, if you install PostgreSQL 17, the configuration files are stored in the ```/etc/postgresql/17/main``` directory.

### Allow connection to primary server from standby server

PostgreSQL supports multiple client authentication methods. In Ubuntu, ```peer``` is the default authentication method used for local connections, while ```scram-sha-256``` is the default for ```host``` connections (this used to be ```md5``` until Ubuntu 21.10).

The following guide assumes that you wish to enable TCP/IP connections and use the ```trust``` method for client authentication. 

Configure the file ```/etc/postgresql/*/main/pg_hba.conf``` to allow connections from your standby server. Since this is within a trusted local network, we'll use ```host``` and ```trust``` authentication for simplicity.

* Edit pg_hba.conf:

```bash
sudo nano /etc/postgresql/17/main/pg_hba.conf
```

* Add the following line for your standby server, replacing ```192.168.1.0/24``` with the IP address range of your replication server's network:

```bash
host    replication       postgres        192.168.1.0/24        trust
```

{hint style="info"}
**Important Considerations**

***Security***: trust authentication bypasses password checks. It is generally suitable only for trusted local networks where access to the server is already restricted. For production environments or less trusted networks, use stronger authentication methods like ```scram-sha-256``` combined with SSL.

***Specificity***: Replace ```192.168.1.0/24``` with the most specific IP range necessary to access the server. Don't use wider ranges than you need to minimize the potential impact of any security vulnerabilities. If possible, restrict access to only the exact IP address of the replication server (i.e. ```192.168.1.58/32```)
{endhint}

### Configure primary server replication

By default, only connections from the local system are allowed. To enable our replication server to connect to our primary server, edit the file ```/etc/postgresql/*/main/postgresql.conf```. Locate the line: ```#listen_addresses = ‘localhost’``` and change it to ```*```:

```bash
sudo nano /etc/postgresql/17/main/postgresql.conf
```

* Uncomment and edit, or add the following lines:

```bash
listen_addresses = '*'
```

{hint style="info"}
**Note:**
```‘*’``` will allow all available IP interfaces (IPv4 and IPv6), to only listen for IPv4 set ```0.0.0.0``` while ```‘::’``` allows listening for all IPv6 addresses.

**For optimal security, only allow the IP address of the replication server.**
{% endhint %}

* To enable streaming replication, uncomment or add the following line:

```bash
wal_level = replica
```

* To enable ***asyncronous*** streaming replication, uncomment and edit or add the following line:

```bash
synchronous_standby_names = '*'
```

* Restart the PostgreSQL service to initialize the new configuration:

```bash
sudo systemctl restart postgresql
```

### Configure primary server replication slot

Each replication slot has a name, which can contain lower-case letters, numbers, and the underscore character.

Existing replication slots and their state can be seen in the [pg_replication_slots](https://www.postgresql.org/docs/current/view-pg-replication-slots.html) view.

* Create replication slot (we will use ```node_a_slot``` as the name)

```bash
sudo -u postgres psql -c "SELECT pg_create_physical_replication_slot('node_a_slot');"
```

* **Example** of expected output:

```
 pg_create_physical_replication_slot
-------------------------------------
 (node_a_slot,)
(1 row)
```

* Confirm slot creation

```bash
sudo -u postgres psql -c "SELECT slot_name, slot_type, active FROM pg_replication_slots;"
```

* **Example** of expected output:

```
  slot_name   | slot_type | active
--------------+-----------+--------
 node_a_slot  | physical  | f
(1 row)
```

## Standby server configuration

* Now, in the **```standby```** server, let’s stop the PostgreSQL service:

```bash
sudo systemctl stop postgresql
```

### Backup the current state of the main server

* Log in as ```postgres``` user

```bash
sudo su - postgres
```

* Remove all content in the data directory

```bash
rm -rf /var/lib/postgresql/17/main/*
```

* Use ```pg_basebackup``` to perform a full single pass of the primary server, copying the content of the main database onto the standby server. In the ```pg_basebackup``` command the flags represent the following:

    **```-h```** : The hostname or IP address of the main server

    **```-D```** : The data directory

    **```-U```** : The user to be used in the operation

    **```-P```** : Turns on progress reporting

    **```-v```** : Enables verbose mode

    **```-R```** : Creates a **```standby.signal```** file and appends connection settings to **```postgresql.auto.conf```**

```bash
pg_basebackup -h <IP address of the main server> -D /var/lib/postgresql/17/main -U postgres -P -v -R
```

<!--- ADD EXAMPLE OF EXPECTED OUTPUT--->
* Exit ```postgres``` user

```bash
exit
```

* Edit postgresql.conf to use previously created replication slot

```bash
sudo nano /etc/postgresql/17/main/postgresql.conf
```

* Add the following line:

```bash
primary_slot_name = 'deadhorse_slot'
```

* Start the PostgreSQL service on the standby server:

```bash
sudo systemctl start postgresql
```

## Verification

* From the primary server, log in as ```postgres``` user and enter query prompt:

```bash
sudo -u postgres psql
```

* Toggle display mode for query results (for narrow displays):

```bash
\x
```

* Query replication status

```bash
SELECT * FROM pg_stat_replication;
```

**Example** of expected output:

```
-[ RECORD 1 ]----+------------------------------
pid              | 1513562
usesysid         | 10
usename          | postgres
application_name | 17/main
client_addr      | 192.168.1.21
client_hostname  |
client_port      | 60638
backend_start    | 2025-01-30 06:15:36.882456+00
backend_xmin     |
state            | streaming
sent_lsn         | 21/5D9A6FD8
write_lsn        | 21/5D9A6FD8
flush_lsn        | 21/5D9A6FD8
replay_lsn       | 21/5D9A6FD8
write_lag        | 00:00:00.001008
flush_lag        | 00:00:00.001421
replay_lag       | 00:00:00.001427
sync_priority    | 1
sync_state       | sync
reply_time       | 2025-02-01 16:28:13.157009+00
```

* Confirm use of replication slot

```bash
SELECT slot_name, slot_type, active FROM pg_replication_slots;
```

**Example** of expected output:

```
-[ RECORD 1 ]-------------
slot_name | node_a_slot
slot_type | physical
active    | t
```

* Quit ```psql```

```bash
\q
```

* Exit ```postgres``` user

```bash
exit
```

### Additional verification

From each server, issue the following command to verify current WAL segment stream. If performed on both servers quickly, you can verify the same WAL segment.

```bash
ps aux | grep postgres
```

**Example** from Primary

```
postgres: 17/main: logical replication launcher
postgres: 17/main: walsender postgres 192.168.1.58(60638) streaming 21/6D2435B0
```

**Example** from Standby

```
postgres: 17/main: walreceiver streaming 21/6D2435B0
```