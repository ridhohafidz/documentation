# PostgreSQL 12 Installation #


First of all:
```bash
$ sudo apt update && sudo apt -y upgrade
```

Add PostgreSQL 12 repository:
```bash
$ sudo apt -y install gnupg2
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```

Add PostgreSQL repository:
```bash
$ echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list
```

Install the PostrgeSQL:
```bash
$ sudo apt-get install postgres
```
![Install postgres](../images/postgre_12_installation.png)

As you can see, `sudo apt-get install postgres` will install additional packages. Depends on your need, you may not want to install those packages. In that case, you can install specific `postgres` packages manually like:
```bash
$ sudo apt-get install postgresql-12 postgresql-client-12
```