**作成日付:2023/12/29**

# 仮想マシン上に Postgresql 環境を構築

## 公式サイト

- https://www.postgresql.org/download/linux/ubuntu/

_以降、仮想マシン上で root 権限で実行_

## postgresql をインストール

```shell
$ sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
$ cat /etc/apt/sources.list.d/pgdg.list
deb http://apt.postgresql.org/pub/repos/apt jammy-pgdg main
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
OK
$ apt update
〜
$ apt -y install postgresql-15
〜
```

## サービスを起動する

```shell
$ systemctl enable postgresql
Synchronizing state of postgresql.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable postgresql
$ systemctl start postgresql
$ systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Sun 2023-12-03 09:25:28 UTC; 5min ago
   Main PID: 3599 (code=exited, status=0/SUCCESS)

Dec 03 09:25:28 postgresql systemd[1]: Starting PostgreSQL RDBMS...
Dec 03 09:25:28 postgresql systemd[1]: Finished PostgreSQL RDBMS.
```

## ユーザを作成する

```shell
$ vi /etc/postgresql/15/main/pg_hba.conf
=====
#local   all             all                                     peer
local   all             all                                     password
=====
$ systemctl restart postgresql
$ sudo -u postgres psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE ROLE readonly;
CREATE ROLE
postgres=# GRANT pg_read_all_data TO readonly;
GRANT ROLE
postgres=# CREATE ROLE readwrite;
CREATE ROLE
postgres=# GRANT pg_read_all_data, pg_write_all_data TO readwrite;
GRANT ROLE
postgres=# CREATE USER bravog WITH CREATEDB ENCRYPTED PASSWORD 'bravog'
postgres=# psql -U bravog -d postgres
postgres-# CREATE DATABASE default
postgres-# \q
```

## 外部に公開する

```shell
$ vi /etc/postgresql/15/main/postgresql.conf
=====
#listen_addresses = 'localhost'         # what IP address(es) to listen on;
listen_addresses = '*'
=====
$ vi /etc/postgresql/15/main/pg_hba.conf
=====
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
host    all             all             0.0.0.0/0               md5
=====
$ service postgresql restart
```

<<< Firewall 利用時のみ >>>

```shell
$ ufw status | grep '5432'
$ ufw allow 5432
$ ufw status | grep '5432'
5432                       ALLOW       Anywhere
5432 (v6)                  ALLOW       Anywhere (v6)
```
