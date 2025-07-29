---
layout: post
title: "Migrating from sqlite to postgresql"
date: 2025-07-28 20:43:00 -0000
categories: runbooks
---

```bash
#Attempt to try this in a lab first before trying on prod setup

Spin up Prometheus and Grafana instance, ingest from test device x
Test that you're getting metrics from the test device x

WARNING: This is purely based on my own notes, and might not be completely correct/factual enough to be used in a real enterprise environment.
```

1. Migrating Grafana from sqlite to postgresql db
    1. Make a backup of the current Grafana sqlite db
    2. Install Grafana into a container (this is a dummy Grafana instance used to setup PostgreSQL with the correct schema)
    3. Install Postgres into a container (this is the Postgres database that will be dumped and migrated into once it has obtained the correct schema)
    4. Create Grafana database inside the Postgres instance
    5. Restart grafana container and dump database to a specified location
    6. Use pgloader to migrate data from sqlite to postgressql
    7. Check data has migrated over successfully and run some tests
    8. Point current instance of Grafana to new postgresql instance
    9. Check that everything works
    10. Shut down/cleanup any unnecessary resources
        
        ```bash
        #try with same grafana version in a testing environment (in case the schema changes)
        #
        
        #stop grafana and make copy of grafana sqlite db and copy to own workstation
        
        systemctl stop grafana-server
        
        cp /usr/local/grafana/data/grafana.db /root/grafana.db.backup 
        
        # make sure you can ssh in the first place
        scp root@<grafana-lxc-ip>:/root/grafana.db.backup ./grafana.db
        
        #install postgres into container 
        
        setup the lxc container with networking 
        
        apt update && apt install -y postgresql postgresql-client
        
        vim /etc/postgresql/15/main/pg_hba.conf
        host    grafana     grafana     <grafana-ip>/32     md5
        
        vim /etc/postgresql/15/main/postgresql.conf
        listen_addresses = '*'
        
        systemctl restart postgresql
        
        #create grafana database and user in postgresql
        
        su -c /usr/bin/psql postgres
        CREATE USER grafana WITH PASSWORD 'grafana_pass';
        CREATE DATABASE grafana WITH OWNER=grafana ENCODING 'UTF8' TEMPLATE template0;
        \q
        
        #install grafana into container
        
        wget https://dl.grafana.com/enterprise/release/grafana-enterprise-12.0.2.linux-amd64.tar.gz
        tar -zxvf grafana-enterprise-12.0.2.linux-amd64.tar.gz
        
        useradd -r --no-create-home --shell /usr/sbin/nologin grafana #added the -r flag but didn't add it before
        
        mv grafana-v12.0.2 /usr/local/grafana
        
        chown -R grafana:grafana /usr/local/grafana
        
        cp /usr/local/grafana/conf/sample.ini /usr/local/grafana/conf/grafana.ini
        
        chown grafana:grafana /usr/local/grafana/conf/grafana.ini
        chmod 640 /usr/local/grafana/conf/grafana.ini
        
        also edit the /usr/local/grafana/conf/grafana.ini to include:
        [database]
        type = postgres
        host = <postgres-lxc-ip>:5432
        name = grafana
        user = grafana
        password = grafana_pass
        ssl_mode = disable
        
        touch /etc/systemd/system/grafana-server.service
        
        [Unit]
        Description=Grafana Server
        Wants=network-online.target
        After=network-online.target
        
        [Service]
        User=grafana
        Group=grafana
        ExecStart=/usr/local/grafana/bin/grafana-server \
          --homepath=/usr/local/grafana
        Restart=on-failure
        
        [Install]
        WantedBy=multi-user.target
        
        also edit the grafana.ini to include:
        [database]
        type = postgres
        host = <postgres-lxc-ip>:5432
        name = grafana
        user = grafana
        password = grafana_pass
        
        systemctl daemon-reload
        systemctl enable --now grafana-server
        
        #use pgloader to migrate from sqlite to postgresql
        
        brew install pgloader
        
        create a file called grafana.load:
        
        LOAD DATABASE
        FROM sqlite:///Users/hassan/Work/grafana.db
        INTO postgresql://grafana:squid@192.168.8.157:5432/grafana
        WITH truncate, create no tables, create indexes, reset sequences
        SET work_mem to '16MB', maintenance_work_mem to '512 MB';
        
        pgloader grafana.load
        
        Checking state of the db:
        get the schema of both the sqlite db and the postgresql db and compare
        get the number of rows and compare
        check null columns count
        check boolean column values
        
        #point current instance of Grafana to postgresql instance

        (this should be via a custom.ini file in /usr/local/grafana/conf
        change:
        [database]
        type = postgres
        host = <postgres-lxc-ip>:5432
        name = grafana
        user = grafana
        password = grafana_pass
        
        systemctl restart grafana-server

        Also update the pg_hba.conf file to include the IP of the prod grafana host if you haven't already
        
        #check that everything works
        
        systemctl status grafana-server
        
        journalctl -u grafana-server --since "5 minutes ago"
        
        curl -s -u admin:admin http://<grafana-ip>:3000/api/health
        
        curl -u admin:admin http://localhost:3000/api/search
        
        log into grafana ui
        add a dashboard and check it works (can also check the db for activity in the relevant table, and established connections on the            grafana node to postgresql)
        check logs

        nice to haves:
        find a way of doing it programatically 
        
        ```
