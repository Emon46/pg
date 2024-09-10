# pg

// for major upgrade need to take few steps one by one. make sure this process run  one after an other. other wise there will be failure scenario.
// 0. we have previously copied old binary. in the main upgrade func
// 1. transfer primary role to pod-0
// 2. pause postgres sidecar `pg-coordinator` so that the sidecar don't apply any change when we are doing other jobs from ops request. we need to make sure that the sidecar is paused. Or, we may face data loss issue.
// 3. restart pod-0
// 4. then we need to initialize with initdb a new directory where we are going to store updated data from old data directory. here new directory is /var/pv/data.new .
// 5. then make sure the server that was running with old data directory is shutdown gracefully. if not run it in single user mode and shut it down gracefully
// 6. apply pg_upgrade
// 7. mode data from new directory to old directory .
// 8. transfer leader role to pod-0
// 9. resume coordinator sidecar
// 10. then restart other pods
// finally update the condition UpdateStatefulSetImage


```
export PGDATA=/var/pv/data.new && rm -rf /var/pv/data.new/* && /scripts/initdb.sh
```
```
cp -r /var/pv/old-share-dir/* $(/var/pv/old-bin/pg_config --sharedir); cp -r /var/pv/old-lib-dir/* $(/var/pv/old-bin/pg_config --pkglibdir);
```
```
/var/pv/old-bin/postgres --single -O -D /var/pv/data template1
```
```
pg_upgrade --username=postgres --old-port=6380 --new-port=6381 --old-bindir=/var/pv/old-bin/ --new-bindir=$(pg_config --bindir) --old-datadir=/var/pv/data  --new-datadir=/var/pv/data.new --link
```
