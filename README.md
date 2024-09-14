# pg_upgrade test run

## setup environment for testing


copy the data directory:

```
cp -R /var/opt/gitlab/postgresql/data /home/hemon-hkg/pg-data-em-test/ &
```

change the ownere-ship to use `gitlab-psql` 
```
chown gitlab-psql:gitlab-psql -R /home/hemon-hkg/pg-data-em-test/data/* && \
            chown gitlab-psql:root /home/hemon-hkg/pg-data-em-test/data
```


## main testing

### Exec into docker file. Please change the `MOUNT_DIR` to `/var/opt/gitlab/postgresql/` for original run.

``` 
export MOUNT_DIR=/home/hemon-hkg/pg-data-em-test/ && \
        mkdir $MOUNT_DIR/pg-upg-logs && chmod 0777 $MOUNT_DIR/pg-upg-logs && \
        docker run -it -v $MOUNT_DIR/:/var/opt/gitlab/postgresql/ \
        -v $MOUNT_DIR/pg-upg-logs:/var/log/gitlab/postgresql \
        -v /opt/gitlab/embedded/ssl/certs:/opt/gitlab/embedded/ssl/certs \
        --entrypoint bash  ghcr.io/zalando/spilo-15:3.0-p1
```

### create a user called `gitlab-psql` this is the super user used in gitlab postgres and this user will be used for rest of the upgrade process

``` 
groupadd -g 986 gitlab-psql && \
            useradd -u 991 -g 986 -d /var/opt/gitlab/postgresql -s /bin/sh gitlab-psql
```

### create a new data directory for upgraded 14 version data to store.
``` 
mkdir /var/opt/gitlab/postgresql/data-new  && chmod 0750 /var/opt/gitlab/postgresql/data-new && \
                                                chown gitlab-psql:root /var/opt/gitlab/postgresql/data-new
```

### create a new directory to keep storing the old conf files for postgresql. Incase of disaster we can recover conf from this file
``` 
mkdir /var/opt/gitlab/postgresql/conf-old && chmod 0777 /var/opt/gitlab/postgresql/conf-old
```

``` 
su - gitlab-psql
cp /var/opt/gitlab/postgresql/data/postgresql.conf \
    /var/opt/gitlab/postgresql/data/runtime.conf \
    /var/opt/gitlab/postgresql/data/pg_hba.conf \
    /var/opt/gitlab/postgresql/data/pg_ident.conf \
    /var/opt/gitlab/postgresql/data/postgresql.base.conf \
    /var/opt/gitlab/postgresql/conf-old/
```

###  initialize new repository

```
/usr/lib/postgresql/14/bin/initdb --pgdata=/var/opt/gitlab/postgresql/data-new --encoding=UTF8 --locale=C  -U gitlab-psql 
```

###  update local socket auth to trust to make the process easy 
``` 
sed -i 's/peer map=gitlab/trust/'  /var/opt/gitlab/postgresql/data/pg_hba.conf
```

### remove wal-g config for the time of upgrade process only
``` 
export CONF_PATH="/var/opt/gitlab/postgresql/data/runtime.conf" && \
    sed -i "/archive_command = '\/opt\/wal-g\/archive-walg.sh %p'/d" "$CONF_PATH" && \
    sed -i '/archive_timeout = 120/d' "$CONF_PATH" && \
    sed -i '/# number of seconds; 0 disables/d' "$CONF_PATH"
```

### copy the conf from old directory to new directory
```
cp /var/opt/gitlab/postgresql/data/postgresql.conf \
    /var/opt/gitlab/postgresql/data/runtime.conf \
    /var/opt/gitlab/postgresql/data/pg_hba.conf \
    /var/opt/gitlab/postgresql/data/pg_ident.conf \
    /var/opt/gitlab/postgresql/data/postgresql.base.conf \
    /var/opt/gitlab/postgresql/data/server.key \
    /var/opt/gitlab/postgresql/data/server.crt /var/opt/gitlab/postgresql/data-new/
```

### If the shut down was not happened gracefully we can start it in single user mode. it should recover and there will be a `backend>` when recover is done. this time the service is not accessable by any user

``` 
/usr/lib/postgresql/13/bin/postgres --single -D /var/opt/gitlab/postgresql/data template1
```

### run upgrade after that 
```
/usr/lib/postgresql/14/bin/pg_upgrade --jobs=$(( $(grep -c -E '^processor' /proc/cpuinfo) - 1 )) \
    -U gitlab-psql \
    -b /usr/lib/postgresql/13/bin/ \
    -B /usr/lib/postgresql/14/bin/ \
    -d /var/opt/gitlab/postgresql/data \
    -D /var/opt/gitlab/postgresql/data-new --link 

```

### start the new server now 

``` 
/usr/lib/postgresql/14/bin/pg_ctl -D /var/opt/gitlab/postgresql/data-new start \
                                    -U gitlab-psql \
                                    -o "-p 5433 -c unix_socket_directories='/var/opt/gitlab/postgresql'"
```


### check the status of the server
``` 
/usr/lib/postgresql/14/bin/pg_ctl status -D /var/opt/gitlab/postgresql/data-new
```
### run the vaccum annalyzer 
``` 
/usr/lib/postgresql/14/bin/vacuumdb \
    -U gitlab-psql \
    --all \
    --analyze-in-stages \
    -p 5433 -j $(( $(grep -c -E '^processor' /proc/cpuinfo) - 1 )) \
    --echo -h /var/opt/gitlab/postgresql
```

### create the extensions

``` 
/usr/lib/postgresql/14/bin/psql -U gitlab-psql \
    -d gitlabhq_production -p 5433 \
    -h /var/opt/gitlab/postgresql \
    -f update_extensions.sql
```

## If you are here that means the upgrade was successful!!!!!!!!!!!!


# miscellaneous


```
$ cat delete_old_cluster.sh
#!/bin/sh

rm -rf '/var/opt/gitlab/postgresql/data'
```

```
$ cat update_extensions.sql
\connect gitlabhq_production
ALTER EXTENSION "btree_gist" UPDATE;
ALTER EXTENSION "pg_stat_statements" UPDATE;
ALTER EXTENSION "pg_trgm" UPDATE;
```


some of the important files
``` 
/opt/gitlab/embedded/bin/pg_controldata -D /var/opt/gitlab/postgresql/data/

cat /var/opt/gitlab/postgresql/pgpass

cat /etc/gitlab/gitlab.rb
```







# pg upgrade design

### Patroni with postgres
- diable write mode in leader node
- pause patroni with api call https://patroni.readthedocs.io/en/latest/pause.html with this there will be no unintentional failover
- Shutdown database (or Postgres service).
- run initdb to initialize new database
  ```
  export PGDATA=/var/pv/data.new && rm -rf /var/pv/data.new/* && /<pg-newVersion>/initdb --pgdata=/var/pv/data.new
  ```
- Copy configurations file pg_hba.conf, postgres.conf, postgres_auto.cong or other files if there is any.
- do pg_upgrade (check the log properly, if there is any post action required it should be detailed in the logs)
- fix patroni configuration
- DCS (Dynamic Configuration Settings) check how we can do it
- start patroni



### Interesting insite with update procedure

this copy is for backup of the data. incase something went wrong we can do in place recovery. without any down time. The idea is just replace the directory and restart postgres
```
cp -r /var/opt/gitlab/postgresql/data /home/hemon-hkg/pg-data-em-test/ &
```


```
  /opt/gitlab/embedded/bin/pg_controldata -D /var/opt/gitlab/postgresql/data/
```

copy this 4 files:
```
  /var/opt/gitlab/postgresql/data/postgresql.conf
  /var/opt/gitlab/postgresql/data/runtime.conf
  /var/opt/gitlab/postgresql/data/pg_hba.conf
  /var/opt/gitlab/postgresql/data/pg_ident.conf
```



```
cp /var/opt/gitlab/postgresql/data/postgresql.conf /var/opt/gitlab/postgresql/data/runtime.conf /var/opt/gitlab/postgresql/data/pg_hba.conf /var/opt/gitlab/postgresql/data/pg_ident.conf /var/opt/gitlab/postgresql/data/postgresql.base.conf ./conf-old/
```

remove archiving section from /var/opt/gitlab/postgresql/data/runtime.conf
```
# - Archiving -
archive_command = '/opt/wal-g/archive-walg.sh %p'   # command to use to archive a logfile segment
archive_timeout = 120    # force a logfile segment switch after this
        # number of seconds; 0 disables
```

```
sed -i 's/peer map=gitlab/trust/'  pg_hba.conf
```

```
export CONF_PATH="runtime.conf" && sed -i "/archive_command = '\/opt\/wal-g\/archive-walg.sh %p'/d" "$CONF_PATH" && sed -i '/archive_timeout = 120/d' "$CONF_PATH" && sed -i '/# number of seconds; 0 disables/d' "$CONF_PATH"

```
create a home dir for internal upgrade new version. this one is temporary. no consitent data require. just to put the logs intermediately
```
mkdir /home/hemon-hkg/pg-data-em-test/hpg && chmod /home/hemon-hkg/pg-data-em-test/hpg
```

docker run to have both binaries:
```
docker run -it -v /home/hemon-hkg/pg-data-em-test/:/var/opt/gitlab/postgresql/ -v /home/hemon-hkg/pg-data-em-test/logs:/var/log/gitlab/postgresql -v /opt/gitlab/embedded/ssl/certs:/opt/gitlab/embedded/ssl/certs --entrypoint bash  ghcr.io/zalando/spilo-15:3.0-p1
```

```
# docker run -it -v /home/hemon-hkg/pg-data-em-test/:/var/opt/gitlab/postgresql/ -v /home/hemon-hkg/pg-data-em-test/logs:/var/log/gitlab/postgresql -v /opt/gitlab/embedded/ssl/certs:/opt/gitlab/embedded/ssl/certs --entrypoint bash  ghcr.io/zalando/spilo-15:3.0-p1


docker run -it -v /home/hemon-hkg/pg-data-em-test/:/var/opt/gitlab/postgresql/ \
               -v /home/hemon-hkg/pg-data-em-test/logs:/var/log/gitlab/postgresql \
               -v /opt/gitlab/embedded/ssl/certs:/opt/gitlab/embedded/ssl/certs \
               --entrypoint bash  ghcr.io/zalando/spilo-15:3.0-p1

groupadd -g 986 gitlab-psql &&     useradd -u 991 -g 986 -d /var/opt/gitlab/postgresql -s /bin/sh gitlab-psql

su - gitlab-psql



mkdir /var/opt/gitlab/postgresql/conf-old && cp /var/opt/gitlab/postgresql/data/postgresql.conf /var/opt/gitlab/postgresql/data/runtime.conf /var/opt/gitlab/postgresql/data/pg_hba.conf /var/opt/gitlab/postgresql/data/pg_ident.conf /var/opt/gitlab/postgresql/data/postgresql.base.conf /var/opt/gitlab/postgresql/conf-old/


mkdir /var/opt/gitlab/postgresql/data-new
/usr/lib/postgresql/14/bin/initdb --pgdata=/var/opt/gitlab/postgresql/data-new --encoding=UTF8 --locale=C  -U gitlab-psql


sed -i 's/peer map=gitlab/trust/'  /var/opt/gitlab/postgresql/data/pg_hba.conf

export CONF_PATH="/var/opt/gitlab/postgresql/data/runtime.conf" && sed -i "/archive_command = '\/opt\/wal-g\/archive-walg.sh %p'/d" "$CONF_PATH" && sed -i '/archive_timeout = 120/d' "$CONF_PATH" && sed -i '/# number of seconds; 0 disables/d' "$CONF_PATH"

cp /var/opt/gitlab/postgresql/data/postgresql.conf /var/opt/gitlab/postgresql/data/runtime.conf /var/opt/gitlab/postgresql/data/pg_hba.conf /var/opt/gitlab/postgresql/data/pg_ident.conf /var/opt/gitlab/postgresql/data/postgresql.base.conf /var/opt/gitlab/postgresql/data-new/



/usr/lib/postgresql/14/bin/pg_upgrade --jobs=$(( $(grep -c -E '^processor' /proc/cpuinfo) - 1 )) -U gitlab-psql -b /usr/lib/postgresql/13/bin/ -B /usr/lib/postgresql/14/bin/ -d /var/opt/gitlab/postgresql/data -D /var/opt/gitlab/postgresql/data-new --link


/usr/lib/postgresql/14/bin/pg_ctl -D /var/opt/gitlab/postgresql/data-new start -U gitlab-psql -o "-p 5433 -c unix_socket_directories='/var/opt/gitlab/postgresql'"


/usr/lib/postgresql/14/bin/pg_ctl status -D /var/opt/gitlab/postgresql/data-new


/usr/lib/postgresql/14/bin/vacuumdb -U gitlab-psql --all --analyze-in-stages -p 5433 -j $(( $(grep -c -E '^processor' /proc/cpuinfo) - 1 )) --echo -h /var/opt/gitlab/postgresql


/usr/lib/postgresql/14/bin/psql -U gitlab-psql -d gitlabhq_production -p 5433 -h /var/opt/gitlab/postgresql -f update_extensions.sql

```


init new data dir:
```
su - postgres -c "mkdir /var/opt/gitlab/postgresql/data-new"
```

single user mode incase we need to change anything in database:

```
su - postgres -c "/usr/lib/postgresql/13/bin/postgres --single -D /var/opt/gitlab/postgresql/data template1"

cltr+d
```

Initidb with new version:
```
su - postgres -c "/usr/lib/postgresql/14/bin/initdb --pgdata=/var/opt/gitlab/postgresql/data-new --encoding=UTF8 --locale=C  -U gitlab-psql"
```


run pg_upgrade
```
su - postgres -c "/usr/lib/postgresql/14/bin/pg_upgrade --jobs=3 -U gitlab-psql -b /usr/lib/postgresql/13/bin/ -B /usr/lib/postgresql/14/bin/ -d /var/opt/gitlab/postgresql/data -D /var/opt/gitlab/postgresql/data-new --link"
```

# after upgrade

```
/usr/lib/postgresql/14/bin/pg_ctl -D /var/opt/gitlab/postgresql/data-new start -U gitlab-psql -o "-p 5433 -c unix_socket_directories='/var/opt/gitlab/postgresql'"

/usr/lib/postgresql/14/bin/pg_ctl status -D /var/opt/gitlab/postgresql/data-new

/usr/lib/postgresql/14/bin/vacuumdb -U gitlab-psql --all --analyze-in-stages -p 5433 -j 5 --echo -h /var/opt/gitlab/postgresql

/usr/lib/postgresql/14/bin/psql -U gitlab-psql -d gitlabhq_production -p 5433 -h /var/opt/gitlab/postgresql -f update_extensions.sql
```


```
CREATE ROLE postgres WITH SUPERUSER LOGIN PASSWORD 'your_password';
```

### k8s cluster update


for major upgrade need to take few steps one by one. make sure this process run  one after an other. other wise there will be failure scenario.
0. we have previously copied old binary. in the main upgrade func

1. transfer primary role to pod-0
2.  pause postgres sidecar `pg-coordinator` so that the sidecar don't apply any change when we are doing other jobs from ops request. we need to make sure that the sidecar is paused. Or, we may face data loss issue.
3. restart pod-0
4. then we need to initialize with initdb a new directory where we are going to store updated data from old data directory. here new directory is /var/pv/data.new .
5. then make sure the server that was running with old data directory is shutdown gracefully. if not, run it in single user mode and shut it down gracefully
6. apply pg_upgrade
7. mode data from new directory to old directory .
8. transfer leader role to pod-0
9. resume coordinator sidecar
10. then restart other pods

finally update the condition UpdateStatefulSetImage


```
export PGDATA=/var/pv/data.new && rm -rf /var/pv/data.new/* && /<pg-newVersion>/initdb --pgdata=/var/pv/data.new
```

```
cp -r /var/pv/old-share-dir/* $(/var/pv/old-bin/pg_config --sharedir); cp -r /var/pv/old-lib-dir/* $(/var/pv/old-bin/pg_config --pkglibdir);
```

### this was for graceful shutdown
```
/var/pv/old-bin/postgres --single -O -D /var/pv/data template1
```

### run upgrade
```
pg_upgrade --username=postgres --old-port=6380 --new-port=6381 --old-bindir=/var/pv/old-bin/ --new-bindir=$(pg_config --bindir) --old-datadir=/var/pv/data  --new-datadir=/var/pv/data.new --link
```
### run the post scripts that generated by pg-upgrade
./analuyze_pg_script.sh
