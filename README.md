# pg_upgrade test run

## analysis
- ubuntu image we used for this test run `ghcr.io/zalando/spilo-15:3.0-p1`
- luckily we don't have seperate tablespace which would have add more complexity 
- can't use anything else but `--locale=C` in initdb as it was used for older cluster
- vaccum binary works faster then usual process. so we should use that one
- it took 5 mnts to setup the docker images and infra
- took 2 minutes to run the initidb 
- it tooks 3-4 minutes to run pg-upgrade process
- it tooks 5-6 minutes to run the post upgrade anylisis
- so if we are super optimize in the whole process we can have atmost 20-25 minutes of down time

## quesitons
- how to update the embedded binary that we have inside `/opt/gitlab/embedded/bin/` with latest 14 binary?
- i have tested this with ubuntu image? need to check if we can generate a centos image which can have both 13 and 14 just like the ubunut image we used
  

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



