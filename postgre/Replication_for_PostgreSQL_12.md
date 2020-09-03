# Streaming Replication for PostgreSQL 12 #
---

Install the PostgreSQL
`sudo apt-get update & sudo apt-get install postgres`

Inititalize and start PostgreSQL on the Master
`sudo systemctl start postgresql@12-main`

Modify `listen_addresses` parameter to allow a specific IP interface or all (using *). Modifying this requires a restart for the change to take effect.
`sudo su - postgres`
`psql -c "ALTER SYSTEM SET listen_addresses TO '*';"`
`exit`
`sudo systemctl restart postgresql`

Create a user for replication in the Master. It is discouraged to use superuser postgres in order to setup replication, though it works.
`sudo su - postgres`
`psql`
`CREATE USER replicator WITH REPLICATION;"`
`\password replicator`
=> chrollolucifer

Allow replication connections from Standby to Master by appending similar line as following to the `pg_hba.conf`:
`host replication replicator  10.119.234.68/32  md5`