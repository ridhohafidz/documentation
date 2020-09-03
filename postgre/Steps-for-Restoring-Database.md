# Steps for Database Restoration
---
Assume we want to replace database `database-1` in `server A` from a `pg_dump` created from the updated version of `database-1` in `server Z`.

Table of contents:
- [Steps in server Z (origin)](#in-server-Z)
- [Steps in server A (destination)](#in-server-A)
---

## In `server Z`:
1. `sudo su - postgres`
2. `pg_dump database-1 > database_1.sql`
3. `scp database_1.sql <your_username_on_server_A>@<server_A_IP_address>:<destination>`

## In `server A`:
1. `cd <destination>`
2. `sudo cp database-1.sql /var/lib/postgresql/database-1.sql`
3. `sudo su - postgres`
4. `dropdb database-1`
5. `createdb database-1 -O <supposed_owner_of_database>`
6. `psql --single-transaction database-1 < database-1.sql`