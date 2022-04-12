---
title: "Backup MySQL Databases in Kubernetes"
date: 2021-03-03T19:20:27+02:00
tags: ["docker", "kubernetes", "cronjob", "bash", "backup"]
draft: false
---
In this post, we will show you how to create a MySQL server backup using Kubernetes CronJobs.

In our case, we do not have a managed MySQL server. But we want to backup it to our NAS, so that we have a backup in case of emergency.
For this we first build a container that can execute our tasks, because we will certainly need several tasks to backup our cluster.

## CronJob Agent Container
First, we'll show you our Dockerfile so you know what we need.

```Dockerfile
FROM alpine:3.10

# Update
RUN apk --update add --no-cache bash nodejs-current yarn curl busybox-extras vim rsync git mysql-client openssh-client 
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl && chmod +x ./kubectl && mv ./kubectl /usr/local/bin/kubectl

# Scripts
RUN mkdir /srv/jobs
COPY jobs/* /srv/jobs/

# Backup Folder
RUN mkdir /var/backup
RUN mkdir /var/backup/mysql
```

## Backup Script
And now our backup script which the container executes.

Our script is quite simple, we get all tables with the mysql client, export them as sql file, pack them in a zip file and send them in a 8 hours interval to our NAS.

```bash
#!/bin/bash

############# SET VARIABLES #############

# Env Variables
BACKUPSERVER="8.8.8.8" # Backup Server Ip
BACKUPDIR=/var/backup/mysql
BACKUPREMOTEDIR="/mnt/backup/kubernetes/"
HOST="mariadb.default"
NOW="$(date +"%Y-%m-%d")"
STARTTIME=$(date +"%s")
USER=mysqlUser
PASS=mysqlPassword


############# BUILD ENVIROMENT #############
# Check if temp Backup Directory is empty
mkdir $BACKUPDIR

if [ "$(ls -A $BACKUPDIR)" ]; then
    echo "Take action $BACKUPDIR is not Empty"
    rm -f $BACKUPDIR/*.gz
    rm -f $BACKUPDIR/*.mysql
else
    echo "$BACKUPDIR is Empty"
fi

############# BACKUP SQL DATABASES #############
for DB in $(mysql -u$USER -p$PASS -h $HOST -e 'show databases' -s --skip-column-names); do
    mysqldump -u$USER -p$PASS -h $HOST --lock-tables=false $DB > "$BACKUPDIR/$DB.sql";
done

############# ZIP BACKUP #############
cd $BACKUPDIR
tar -zcvf backup-${NOW}.tar.gz *.sql

############# MOVE BACKUP TO REMOTE #############
rsync -avz $BACKUPDIR/backup-${NOW}.tar.gz root@$BACKUPSERVER:$BACKUPREMOTEDIR

# done
```

## Kubernetes CronJob Deployment
Finally we show you the kubernetes deployment for our agent.

In the deployment, our agent is defined as a CronJob that runs every 8 hours.
In addition, we have added an SSH key as a Conifg map so that this can write to the NAS and a certain security is given.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: backup-mariadb
  namespace: default
spec:
  schedule: "0 8 * * *"
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cronjob-agent
              image: xxx/cronjob-agent
              command: ["bash",  "/srv/jobs/backup-mariadb.sh"]
              volumeMounts:
                - mountPath: /root/.ssh/id_rsa.pub
                  name: cronjob-default-config
                  subPath: id_rsa.pub
                - mountPath: /root/.ssh/id_rsa
                  name: cronjob-default-config
                  subPath: id_rsa
                  readOnly: true
                - mountPath: /root/.ssh/config
                  name: cronjob-default-config
                  subPath: config
          volumes:
            - name: cronjob-default-config
              configMap:
                name: cronjob-default-config
                defaultMode: 256
          restartPolicy: Never
```