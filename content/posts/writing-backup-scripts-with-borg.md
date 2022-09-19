---
title: Writing Backup Scripts with Borg
date: 2022-09-19T09:20:27+02:00
tags: ["borg", "linux", "backup"]
draft: false
---

Since we all know that the first rule is "no backup, no pity", I'll show you how you can use Borg to back up your important data in an encrypted way with relative ease.

If you do not want to use a second computer, but an external hard drive, you can adjust this later in the script and ignore the points in the instructions for the second computer.

### Requirements
 - 2 Linux Computers
 - Borg
 - SSH
 - Storage
 - More than 5 brain cells

### Installation
First we need to install borg on both computers so that we can back up on one and save on the other.
```bash
sudo apt install borgbackup
```

Then we create a Borg repository. We can either use an external target or a local path.

**External Target:**
```bash
borg init --encryption=repokey ssh://user@192.168.2.42:22/mnt/backup/borg
```

**Local Path:**
```bash
borg init --encryption=repokey /path/to/backup_folder
```

If you are using an external destination, I recommend that you store your SSH key on the destination.
This way you don't have to enter a password and is simply nicer from my point of view.

Once you have created everything and prepared the script with your parameters, I recommend that you run the script as a CronJob so that you no longer have to remember to back up your things yourself.

**crontab example:**
```bash
#Minute Hour    Day    Month  Day(Week)      command
#(0-59) (0-23)  (1-31)  (1-12)  (1-7;1=Mo)
00 2 * * * /srv/scripts/borgBackup.sh
```

### Automated script
```bash
#!/bin/sh

# VARS
BACKUPSERVER="192.168.2.42"
BACKUPDIR="/mnt/backup/borg"

# Here you can either use your external destination or the local path.
# External target
export BORG_REPO="user@$BACKUPSERVER:$BACKUPDIR"

# Local path
# export BORG_REPO=/path/to/backup_folder

# Your repository password must be stored here.
export BORG_PASSPHRASE='S0m3th1ngV3ryC0mpl1c4t3d'

info() { printf "\n%s %s\n\n" "$( date )" "$*" >&2; }
trap 'echo $( date ) Backup interrupted >&2; exit 2' INT TERM

info "Start backup"

#Here the backup is created, adjust it the way you would like to have it.
borg create                    \
    --stats                    \
    --compression lz4          \
    ::'BackupName-{now}'        \
    /etc/nginx                  \
    /home/user

backup_exit=$?

info "Deleting old backups"
# Automatic deletion of old backups
borg prune                          \
--prefix 'BackupName-'              \
    --keep-daily    7              \
    --keep-weekly  4                \
    --keep-monthly  6

prune_exit=$?

# Information on whether the backup worked.
global_exit=$(( backup_exit > prune_exit ? backup_exit : prune_exit ))

if [ ${global_exit} -eq 0 ]; then
    info "Backup and Prune finished successfully"
elif [ ${global_exit} -eq 1 ]; then
    info "Backup and/or Prune finished with warnings"
else
    info "Backup and/or Prune finished with errors"
fi

exit ${global_exit}
```

### Get your data from the backup
First, we create a temporary directory in which we can mount the backup.

```bash
mkdir /tmp/borg-backup
```

Once our mount point is created, we can mount our backup repo.
At this point you must remember that you can use an external destination or a local path.

```bash
borg mount ssh://user@192.168.2.42/mnt/backup/borg /tmp/borg-backup
```
Once our repo is mounted, we can change into the directory and restore files via **rsync** or **cp**.

### Conclusion
I hope you could understand everything and now secure your shit sensibly. Because without a backup we are all lost!
