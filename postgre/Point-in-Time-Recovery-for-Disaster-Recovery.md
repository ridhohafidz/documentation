# Postgres Disaster Recovery for Debian 9

There are two types of disaster recovery options well-known in PostgreSQL; **Streaming replication for high availability** and **WAL archiving for point in time recovery**. Each has its own benefits.

The goal for **streaming replication** is to provide a failover site that can be promoted as the main one if the main database goes down for any reason; hardware failure, software failure, or even network outage. As such, it helps **keeping the database online, but not recovering accidentally lost data**.

With **WAL archiving**, however, we can recover accidentally lost data, although we can't provide a failover site immediately. With **WAL archiving**, **a copy of the database can be brought back at any point in time** as long as a base backup from before that time and all `WAL` segments needed up till that time are available.

Therefore, setting up both types of disaster recovery gets us the best of both world.

Let's get started.

## **Streaming Replication for High Availability**

You will need to have two hosts with PostgreSQL. Let's say the `master` will be hosted at `server-A` which has IP address of `10.119.234.62` and the `slave` will be hosted at `server-B` which has IP address of `10.119.234.68`. Let's assume both servers have PostgreSQL up and running. We will stop the service on `server-B` to add it as a slave.

### Master Configuration

- Switch to `postgres` user and type `psql` to enter the Postgres interactive terminal

    ```bash
    $ sudo su – postgres
    $ psql
    ```

    If you have more than one instance of Postgres on your machine and want to connect to a specific instance of that Postgres, find out the port where that instance is running and type:

    ```bash
    $ psql -p [port]
    ```

    If there are two instances of Postgres, the first one installed is probably on port 5432 and the second one is on port 5433. Connecting to `psql` without giving a specific port is likely to connect you with the one in port 5432.

- Create a role with replication privileges to perform replication process:

    ```bash
    postgres=# CREATE ROLE replication WITH REPLICATION PASSWORD 'password' LOGIN;
    ```

- Check that the replication user is indeed created by typing `\du`

![Check replication user](../images/check_replication_user.png)

- Exit the `psql` prompt

    ```bash
    postgres=#\q
    ```

- Change directory to `/etc/postgresql/[version]/main/` and add `replication` entry in `pg_hba.conf` to allow connections between `master` and `slave`:


    ```bash
    $ cd /etc/postgresql/9.4/main/

    $ nano pg_hba.conf

    /etc/postgresql/9.4/main/pg_hba.conf:

    ...
    # TYPE  DATABASE        USER            ADDRESS                 METHOD

    # "local" is for Unix domain socket connections only
    local   all             all                                     peer
    # IPv4 local connections:
    host    all             all             127.0.0.1/32            md5
    # IPv6 local connections:
    host    all             all             ::1/128                 md5
    # Allow replication connections from localhost, by a user with the
    # replication privilege.
    #local   replication     postgres                                peer
    #host    replication     postgres        127.0.0.1/32            md5
    #host    replication     postgres        ::1/128                 md5
    host    replication     replication     10.119.234.68/32        md5
    ```

    Save the file and exit
    
- Configures streaming replication setup by modifying `postgresql.conf`:

    ```bash
    $ nano postgresql.conf

    /etc/postgresql/9.4/main/postgresql.conf:

    #-------------------------------
    # CONNECTIONS AND AUTHENTICATION
    # ------------------------------

    # - Connection Settings -

    listen_addresses = '*'

    ....

    #-------------------------------
    # WRITE AHEAD LOG
    #-------------------------------

    # - Settings -

    wal_level = replica
    # wal_level = hot_standby; for versions < 10
    
    #checkpoint_segments = 16; for versions < 10
    ...

    #-------------------------------
    # REPLICATION
    #-------------------------------

    # sets the maximum number of concurrent connections from the standby servers (ideally set to more than the number of current slaves)
    max_wal_senders = 5

    # sets the minimum number of segments of WAL so that they are not deleted before standby consumes them
    wal_keep_segments = 16

    # hot_standby = on; for versions < 10
    ```

   Some of those configuration are optional and are set in case of future events where the `slave` become `master` and vice versa.

- Exit from role `postgres` and restart PostgreSQL service on master for the changes to take effect

    ```bash
    $ exit

    $ sudo service postgresql restart
    ```

### Slave Configuration

In preparation of the possibility of the `slave` becoming `master`, it's a good idea to make the configuration on slave exactly the same as `master`, especially because we already set the `master` for the possibility of becoming `slave`.

```bash
$ sudo su - postgres

$ psql

postgres=# CREATE ROLE replication WITH REPLICATION PASSWORD 'password' LOGIN;

postgres=# \q

$ cd /etc/postgresql/9.4/main/

$ nano pg_hba.conf

/etc/postgresql/9.4/main/pg_hba.conf:

    ...
    # TYPE  DATABASE        USER            ADDRESS                 METHOD

    # "local" is for Unix domain socket connections only
    local   all             all                                     peer
    # IPv4 local connections:
    host    all             all             127.0.0.1/32            md5
    # IPv6 local connections:
    host    all             all             ::1/128                 md5
    # Allow replication connections from localhost, by a user with the
    # replication privilege.
    #local   replication     postgres                                peer
    #host    replication     postgres        127.0.0.1/32            md5
    #host    replication     postgres        ::1/128                 md5
    host    replication     replication     10.119.234.62/32        md5

$ nano postgresql.conf

/etc/postgresql/9.4/main/postgresql.conf:

    #-------------------------------
    # CONNECTIONS AND AUTHENTICATION
    # ------------------------------

    # - Connection Settings -

    listen_addresses = '*'

    ....

    #-------------------------------
    # WRITE AHEAD LOG
    #-------------------------------

    # - Settings -

    wal_level = replica
    # wal_level = hot_standby; for versions < 10
    
    #checkpoint_segments = 16; for versions < 10
    ...

    #-------------------------------
    # REPLICATION
    #-------------------------------

    # sets the maximum number of concurrent connections from the standby servers (ideally set to more than the number of current slaves)
    max_wal_senders = 5

    # sets the minimum number of segments of WAL so that they are not deleted before standby consumes them
    wal_keep_segments = 16

    # hot_standby = on; for versions < 10

```

Now, we need to transfer the data from `master` to `slave`. This can be achieved simply renaming the data directory folder into something different:

```bash

$ sudo su - postgres

$ cd 9.4

$ mv main old-main

$ mkdir main

$ ls
main  old-main

```
With this, we can restore the old data directory back when things don't go as planned.

Now, execute `pg_basebackup` utility:

```bash
pg_basebackup -h 10.119.234.62 -U replication -D /var/lib/postgresql/9.4/main -P -p 5433
```
Give the password we set previously on the `master` when prompted.

Once it's finished, the newly created `main` directory should now be populated. 

Now, we need to prepare a recovery.conf file which will let the slave connect to master. Create a file with the following content in /var/lib/postgresql/9.4/main/recovery.conf:

```bash
standby_mode = 'on'
primary_conninfo = 'host=10.119.234.62 port=5432 user=replication password=password'
trigger_file = '/tmp/failover.trigger'
```

Don't forget to set all privileges correctly on the Postgres data directory. After that, we are all set to start our slave.

```bash
$ chmod -R g-rwx,o-rwx /var/lib/postgresql/9.4/main/ ; chown -R postgres.postgres /var/lib/postgresql/9.4/main/

```
Exit from the role `postgres` and start the `slave`:

```bash
$ exit

$ sudo service postgresql start
```
We can confirm that the replication process has started by going back to `master` and typing:

```bash
$ sudo su - postgres

$ psql

postgres=# select * from pg_stat_replication;
```

![Check replication status](../images/check_replication_status.png)

## **WAL Archiving for Point In Time Recovery (PITR)**

### **Configuration on Master**

- To properly set WAL archiving, several settings in the `postgresql.conf` need to be adjusted:

```bash
# turning on archive_mode will run archive_command each time a WAL segment is completed
archive_mode = on
# archive_command might be anything from simple cp to rsync
archive_command = 'test ! -f postgres@10.119.234.68:/var/lib/postgresql/wal_archive/%f && rsync –avz %p postgres@10.119.234.34:/var/lib/postgresql/wal_archive/%f'
```
Don't forget to actually create folders the scripts are pointing to.
To check whether the archived WAL is actually sent on the slave, you can run this command on the primary database:

```bash
psql -c "select pg_switch_wal();"

# pg_switch_xlog(); for versions < 10
```
`pg_switch_wal` is used to trigger an immediate segment switch, so that we can check that the `archive_command` is indeed working.

### __Configuration on Slave__


Archived WAL segments need a base backup they can be run on. Without a base backup, they are worthless.

`pg_basebackup` takes base backup of PostgreSQL cluster. If the database cluster in the `PITR` host is set to streaming replication, it is best to take the base backup from the `PITR` host to reduce the load on the main database. In our example of two database however, that is not possible. Hence, we'll take the base backup from the main database cluster which is hosted in the `10.119.234.62`.


```bash
pg_basebackup \
    --pg_data=${CR_BASE_BACKUP_DIR}/${CR_LABEL} \
    --label=${CR_LABEL} \
    --wal-method=stream \
    --user=replication \
    --host=10.119.234.33 \
    --write-recovery-conf \
    --format=plain \
    --progress \
    --verbose
```

Assuming that the `PITR` host is the `Slave` server, than the archived WAL will be sent here.

Archived WAL segments are only needed until the next base backup is taken. Deleting WAL segments manually might lead to accidental corruption of the backups, so it's much safer to use `pg_archivecleanup` pointed to the WAL backup folder, referencing the last `successful` backup. Below is an example script for the task:

```bash
# Find base_backup files not older than 3 weeks
# Sort by date
# Use the oldest one as a reference
OLDEST_BASE_BACKUP=$(basename $(find ${CR_WAL_BACKUP_DIR} -iname "*.backup" -mtime -21 -print0 | \
xargs -0 ls -t | \
tail -n 1))

# Find all subfolders
# Execute pg_archivecleanup for each of the subfolders
find $CR_WAL_BACKUP_DIR \
    -type d \
    -exec pg_archivecleanup -d {} $OLDEST_BASE_BACKUP \;
```

 As paths might be needed in several scipts, a good idea will be to put those in a separate file e.g. `backup.conf`.

> Note: Scripts to automate some of the process of WAL archiving all stored in `/var/lib/postgresql/bin/` at the `Slave` server. Tweak the variables as needed. Provide `.pgpass` file in your postgres root directory (`/var/lib/postgresql/`) in order to run the postgres-cr_backup.sh automatically and successfully

### __Recover The Database__

To recover the database before a specific event, fetch a base backup and WAL data you'd like to recover into to the PostgreSQL data directory. That means replacing the PostgreSQL data directory in the `PITR` host with the extracted content of the base backup.

Bringing the database online now would bring the database to the point the backup took place. To recover all of the WAL transactions between the backup time and the time of the event, set up a `recovery.conf`.

Eventhough there is already a `recovery.conf` from the base backup, to actually do a recovery to a specific point in time, new settings should be added:


```bash
standby_mode = 'on'
primary_conninfo = 'host=10.119.234.33 port=5432 user=replication password=password application_name=pg_slave'
trigger_file = '/tmp/postgresql.trigger.5432'

# Add the following settings

# Here is an example of a simple restore command
restore_command = 'cp /var/lib/postgresql/wal_archive/%f "%p"'

recovery_target_time = '2018-01-29 08:00:00'
```

Modify the settings to suit the need of the backup and start the database. It's a good idea to tail the database log to make sure it's restoring as intended.

Review the content of the database before it comes online.

## Further Reading on the Subject

### Streaming replication
- https://severalnines.com/database-blog/how-setup-streaming-replication-high-availability-postgressql-90
- https://blog.raveland.org/post/postgresql_sr/
- https://www.scalingpostgres.com/tutorials/postgresql-streaming-replication/
- https://dinfratechsource.com/2018/11/10/how-to-set-up-master-slave-streaming-replication-on-postgresql/

### Point In Time Recovery Setup
- https://severalnines.com/database-blog/postgresql-replication-disaster-recovery

### PostgreSQL backup and recovery orchestration
- https://www.zimmi.cz/posts/2018/postgresql-backup-and-recovery-orchestration-wal-archiving/
- https://www.zimmi.cz/posts/2018/postgresql-backup-and-recovery-orchestration-recovery/

### Rsync command
- https://www.computerhope.com/unix/rsync.htm

### WAL archiving
- https://www.scalingpostgres.com/tutorials/postgresql-wal-archiving-pg-receivewal/
